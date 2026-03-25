// SPDX-License-Identifier: GPL-2.0-only
// SPDX-FileCopyrightText: Copyright (c) 2021-2024, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
/*
 * imx390.c - imx390 sensor driver
 * Copyright (c) 2020, RidgeRun. All rights reserved.
 *
 */

// #include <nvidia/conftest.h>

#include <linux/slab.h>
#include <linux/uaccess.h>
#include <linux/gpio.h>
#include <linux/module.h>
#include <linux/seq_file.h>
#include <linux/of.h>
#include <linux/of_device.h>
#include <linux/of_gpio.h>

#include <media/tegra_v4l2_camera.h>
#include <media/tegracam_core.h>

/* imx390 - sensor parameters */
#define IMX390_MIN_GAIN         (0)
#define IMX390_MAX_GAIN         (978)
#define IMX390_ANALOG_GAIN_C0   (1024)
#define IMX390_SHIFT_8_BITS     (8)
#define IMX390_MIN_COARSE_EXPOSURE  (1)
#define IMX390_MAX_COARSE_DIFF  (10)
#define IMX390_MASK_LSB_2_BITS  0x0003
#define IMX390_MASK_LSB_8_BITS  0x00ff
#define IMX390_MIN_FRAME_LENGTH (3092)
#define IMX390_MAX_FRAME_LENGTH (0x1FFFF)


static const int imx390_30_fr[] = {
	30,
};

static const struct camera_common_frmfmt imx390_frmfmt[] = {
	{{1920, 1536}, imx390_30_fr, 1, 0, 0}
};
static const struct of_device_id imx390_of_match[] = {
	{.compatible = "nvidia,imx390-silly",},
	{},
};
MODULE_DEVICE_TABLE(of, imx390_of_match);

static const u32 ctrl_cid_list[] = {
	TEGRA_CAMERA_CID_GAIN,
	TEGRA_CAMERA_CID_EXPOSURE,
	TEGRA_CAMERA_CID_FRAME_RATE,
	TEGRA_CAMERA_CID_HDR_EN,
	TEGRA_CAMERA_CID_SENSOR_MODE_ID,
};

struct imx390 {
	struct i2c_client *i2c_client;
	struct v4l2_subdev *subdev;
	u16 fine_integ_time;
	u32 frame_length;
	struct camera_common_data *s_data;
	struct tegracam_device *tc_dev;
};

static const struct regmap_config sensor_regmap_config = {
	.reg_bits = 16,
	.val_bits = 8,
	.cache_type = REGCACHE_NONE,
};

static int imx390_set_group_hold(struct tegracam_device *tc_dev, bool val)
{
	return 0;
}

static int imx390_set_gain(struct tegracam_device *tc_dev, s64 val)
{
	return 0;
}

static int imx390_set_frame_rate(struct tegracam_device *tc_dev, s64 val)
{
	struct imx390 *priv = (struct imx390 *)tc_dev->priv;

	priv->frame_length = IMX390_MIN_FRAME_LENGTH;
	return 0;
}

static int imx390_set_exposure(struct tegracam_device *tc_dev, s64 val)
{
	return 0;
}

static struct tegracam_ctrl_ops imx390_ctrl_ops = {
	.numctrls = ARRAY_SIZE(ctrl_cid_list),
	.ctrl_cid_list = ctrl_cid_list,
	.set_gain = imx390_set_gain,
	.set_exposure = imx390_set_exposure,
	.set_frame_rate = imx390_set_frame_rate,
	.set_group_hold = imx390_set_group_hold,
};

static int imx390_power_on(struct camera_common_data *s_data)
{
	struct camera_common_power_rail *pw = s_data->power;
	pw->state = SWITCH_ON;
	return 0;
}

static int imx390_power_off(struct camera_common_data *s_data)
{
	struct camera_common_power_rail *pw = s_data->power;
	pw->state = SWITCH_OFF;
	return 0;
}

static int imx390_power_put(struct tegracam_device *tc_dev)
{
	return 0;
}

static int imx390_power_get(struct tegracam_device *tc_dev)
{
	struct device *dev = tc_dev->dev;
	struct camera_common_data *s_data = tc_dev->s_data;
	struct camera_common_power_rail *pw = s_data->power;
	pw->state = SWITCH_ON;
	return 0;
}

static struct camera_common_pdata *imx390_parse_dt(struct tegracam_device
		*tc_dev)
{
	struct device *dev = tc_dev->dev;
	struct device_node *np = dev->of_node;
	struct camera_common_pdata *board_priv_pdata;
	const struct of_device_id *match;
	int err = 0;

	if (!np)
		return NULL;

	match = of_match_device(imx390_of_match, dev);
	if (!match) {
		dev_err(dev, "Failed to find matching dt id\n");
		return NULL;
	}

	board_priv_pdata = devm_kzalloc(dev,
			sizeof(*board_priv_pdata), GFP_KERNEL);
	if (!board_priv_pdata)
		return NULL;

	err = of_property_read_string(np, "mclk", &board_priv_pdata->mclk_name);
	if (err)
		dev_dbg(dev, "mclk name not present, "
				"assume sensor driven externally\n");

	err = of_property_read_string(np, "avdd-reg",
			&board_priv_pdata->regulators.avdd);
	err |= of_property_read_string(np, "iovdd-reg",
			&board_priv_pdata->regulators.iovdd);
	err |= of_property_read_string(np, "dvdd-reg",
			&board_priv_pdata->regulators.dvdd);
	if (err)
		dev_dbg(dev, "avdd, iovdd and/or dvdd reglrs. not present, "
				"assume sensor powered independently\n");

	board_priv_pdata->has_eeprom = of_property_read_bool(np, "has-eeprom");

	return board_priv_pdata;
}

static int imx390_set_mode(struct tegracam_device *tc_dev)
{
	return 0;
}

static int imx390_start_streaming(struct tegracam_device *tc_dev)
{
	struct imx390 *priv = (struct imx390 *)tegracam_get_privdata(tc_dev);

	dev_dbg(tc_dev->dev, "%s:\n", __func__);
	return 0;
}

static int imx390_stop_streaming(struct tegracam_device *tc_dev)
{
	int err;

	err = 0;

	return err;
}

static inline int imx390_read_reg(struct camera_common_data *s_data,
				u16 addr, u8 *val)
{
	return 0;
}

static int imx390_write_reg(struct camera_common_data *s_data,
				u16 addr, u8 val)
{
	return 0;
}

static struct camera_common_sensor_ops imx390_common_ops = {
	.numfrmfmts = ARRAY_SIZE(imx390_frmfmt),
	.frmfmt_table = imx390_frmfmt,
	.power_on = imx390_power_on,
	.power_off = imx390_power_off,
	.write_reg = imx390_write_reg,
	.read_reg = imx390_read_reg,
	.parse_dt = imx390_parse_dt,
	.power_get = imx390_power_get,
	.power_put = imx390_power_put,
	.set_mode = imx390_set_mode,
	.start_streaming = imx390_start_streaming,
	.stop_streaming = imx390_stop_streaming,
};

static int imx390_board_setup(struct imx390 *priv)
{
	struct camera_common_data *s_data = priv->s_data;
	struct device *dev = s_data->dev;
	int err = 0;

	/* Skip mclk enable as this camera has an internal oscillator */

	err = imx390_power_on(s_data);
	if (err)
		dev_err(dev, "error during power on sensor (%d)\n", err);

	return err;
}

static int imx390_open(struct v4l2_subdev *sd, struct v4l2_subdev_fh *fh)
{
	struct i2c_client *client = v4l2_get_subdevdata(sd);

	dev_dbg(&client->dev, "%s:\n", __func__);

	return 0;
}

static const struct v4l2_subdev_internal_ops imx390_subdev_internal_ops = {
	.open = imx390_open,
};

#if defined(NV_I2C_DRIVER_STRUCT_PROBE_WITHOUT_I2C_DEVICE_ID_ARG) /* Linux 6.3 */
static int imx390_probe(struct i2c_client *client)
#else
static int imx390_probe(struct i2c_client *client,
		const struct i2c_device_id *id)
#endif
{
	struct device *dev = &client->dev;
	struct tegracam_device *tc_dev;
	struct imx390 *priv;
	int err;

	dev_dbg(dev, "probing v4l2 sensor at addr 0x%0x\n", client->addr);

	if (!IS_ENABLED(CONFIG_OF) || !client->dev.of_node)
		return -EINVAL;

	priv = devm_kzalloc(dev, sizeof(struct imx390), GFP_KERNEL);
	if (!priv)
		return -ENOMEM;

	tc_dev = devm_kzalloc(dev, sizeof(struct tegracam_device), GFP_KERNEL);
	if (!tc_dev)
		return -ENOMEM;

	priv->i2c_client = tc_dev->client = client;
	tc_dev->dev = dev;
	strncpy(tc_dev->name, "imx390-silly", sizeof(tc_dev->name));
	tc_dev->dev_regmap_config = &sensor_regmap_config;
	tc_dev->sensor_ops = &imx390_common_ops;
	tc_dev->v4l2sd_internal_ops = &imx390_subdev_internal_ops;
	tc_dev->tcctrl_ops = &imx390_ctrl_ops;

	err = tegracam_device_register(tc_dev);
	if (err) {
		dev_err(dev, "tegra camera driver registration failed\n");
		return err;
	}
	priv->tc_dev = tc_dev;
	priv->s_data = tc_dev->s_data;
	priv->subdev = &tc_dev->s_data->subdev;
	tegracam_set_privdata(tc_dev, (void *)priv);

	err = imx390_board_setup(priv);
	if (err) {
		dev_err(dev, "board setup failed\n");
		return err;
	}

	err = tegracam_v4l2subdev_register(tc_dev, true);
	if (err) {
		tegracam_device_unregister(tc_dev);
		dev_err(dev, "tegra camera subdev registration failed\n");
		return err;
	}

	dev_dbg(dev, "detected imx390-silly sensor\n");

	return 0;
}
#define NV_I2C_DRIVER_STRUCT_REMOVE_RETURN_TYPE_INT
#if defined(NV_I2C_DRIVER_STRUCT_REMOVE_RETURN_TYPE_INT) /* Linux 6.1 */
static int imx390_remove(struct i2c_client *client)
#else
static void imx390_remove(struct i2c_client *client)
#endif
{
	struct camera_common_data *s_data = to_camera_common_data(&client->dev);
	struct imx390 *priv = (struct imx390 *)s_data->priv;

	tegracam_v4l2subdev_unregister(priv->tc_dev);
	tegracam_device_unregister(priv->tc_dev);

#if defined(NV_I2C_DRIVER_STRUCT_REMOVE_RETURN_TYPE_INT) /* Linux 6.1 */
	return 0;
#endif
}

static const struct i2c_device_id imx390_id[] = {
	{"imx390-silly", 0},
	{}
};
MODULE_DEVICE_TABLE(i2c, imx390_id);

static struct i2c_driver imx390_silly_i2c_driver = {
	.driver = {
		.name = "imx390-silly",
		.owner = THIS_MODULE,
		.of_match_table = of_match_ptr(imx390_of_match),
	},
	.probe = imx390_probe,
	.remove = imx390_remove,
	.id_table = imx390_id,
};

module_i2c_driver(imx390_silly_i2c_driver);

MODULE_DESCRIPTION("Media Controller driver for IMX390-silly");
MODULE_AUTHOR("NVIDIA Corporation");
MODULE_LICENSE("GPL v2");
MODULE_SOFTDEP("pre: tegra_camera");